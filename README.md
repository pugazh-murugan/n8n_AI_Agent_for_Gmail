# n8n AI Agent for Gmail

A collection of five [n8n](https://n8n.io) workflows that build up from a minimal Gmail AI agent to a multi-tool, voice-enabled assistant. Each workflow is exported as a JSON file that you can import into your own n8n instance.

| # | Workflow | Trigger | Model | What it does |
|---|----------|---------|-------|--------------|
| 1 | [Basic Gmail AI Agent](Basic%20Gmail%20AI%20Agent.json) | n8n chat | `gpt-5-mini` | Chat with an agent that can send email through Gmail |
| 2 | [Unread Email Summariser](Unread%20Email%20Summariser.json) | Manual | `gpt-4o-mini` | Summarises your 5 latest unread emails and emails you the digest |
| 3 | [Telegram Research AI Agent](Telegram%20Research%20Ai%20Agent.json) | Telegram message | `gpt-4.1-mini` | Telegram bot that searches Amazon products via SerpApi |
| 4 | [Multi Tool AI agent](Multi%20Tool%20AI%20agent.json) | Telegram message | `gpt-4o` | Telegram assistant that chains Docs, Calendar, Sheets, Gmail and web search |
| 5 | [Voice AI Agent](Voice%20AI%20Agent.json) | Telegram text or voice | `gpt-4o-mini` | Talk to the agent by voice or text and get a text reply plus a spoken reply |

All workflows are exported with `"active": false`, so nothing runs until you activate it.

---

## Workflows

### 1. Basic Gmail AI Agent

The starting point: the smallest useful agent, with the four building blocks of an n8n AI agent.

```
When chat message received ──▶ Perception (AI Agent)
                                  ├─ Brain    (OpenAI chat model)
                                  ├─ Memory   (window buffer)
                                  └─ Action   (Gmail: send message)
```

- **Trigger:** n8n's built-in chat (`chatTrigger`), which replies with the output of the last node.
- **Gmail tool:** the model fills in `To`, `Subject` and `Message` itself (`$fromAI`), so it can send a mail from a plain-language request.
- **Memory:** a window buffer, so the agent remembers earlier turns of the chat.
- **Note:** the agent has no system prompt, so its behaviour depends entirely on the model and your message.

### 2. Unread Email Summariser

A linear (non-agentic-loop) pipeline that turns your inbox into a short digest.

```
Manual trigger ─▶ Get unread emails ─▶ AI Agent ─▶ Aggregate ─▶ Send summary email
                   (Gmail, limit 5)    (1 line each)  (join)      (HTML numbered list)
```

1. Fetches up to **5 unread** emails from Gmail.
2. For each email, the AI agent writes **one line of at most 20 words**, using the sender, subject and snippet.
3. `Aggregate` collects all the lines into one list.
4. A Gmail node sends the list as an HTML ordered list to a fixed recipient.

**Before you run it:** the recipient address is hard-coded in the `Send summary email` node, and the subject says "5 unread emails" even though the actual count depends on how many unread emails exist. Change both to suit you. To run it on a schedule, replace the manual trigger with a Schedule Trigger.

### 3. Telegram Research AI Agent

```
Telegram Trigger ─▶ Research Agent ─▶ Send a text message
                      ├─ OpenAI Chat Model (gpt-4.1-mini)
                      ├─ Simple Memory (last 2 messages, keyed by Telegram chat id)
                      └─ Amazon search in SerpApi
```

Send the bot a question and it replies in the same chat. The agent's only tool is **Amazon product search via SerpApi** (it can also take a delivery ZIP code), which makes it a shopping and price-research assistant. Memory is limited to a window of 2 and keyed per Telegram chat, so each conversation stays separate.

> This workflow uses the **community node** `n8n-nodes-serpapi`. Install it in your n8n instance (Settings → Community nodes) before importing, or the Amazon tool will show as an unknown node.

### 4. Multi Tool AI agent

The most "agentic" workflow. A Telegram message goes to a GPT-4o agent that is instructed to call **every tool a request needs in the same turn**.

```
Telegram Trigger ─▶ AI Agent ─▶ Send Telegram Reply
                      ├─ OpenAI Chat Model (gpt-4o)
                      ├─ Simple Memory (per Telegram chat id)
                      └─ Tools
                          ├─ Document ............ create a Google Doc, returns its id
                          ├─ Update a document ... write content into that doc
                          ├─ calendar ............ create a Google Calendar event
                          ├─ sheet ............... append a row to Google Sheets
                          ├─ gmail ............... send an email
                          └─ Google search (SerpApi)
```

Example request: *"Schedule a team sync tomorrow at 10, add it to the sheet, write an agenda doc and email it to alex@example.com."*

How the system prompt keeps it reliable:

- **Doc creation is a two-step chain:** `Document` creates an empty doc and returns an `id`; the agent must then pass that exact `id` to `Update a document` and never invent one.
- **Calendar times** are ISO 8601 in `Asia/Kolkata` (e.g. `2026-11-24T10:00:00+05:30`), default duration 1 hour, with today's date injected through `{{ $now }}`.
- **Search** must always get a clear, non-empty query.
- After all tools finish, the agent replies with a short summary that is sent back to Telegram.

**Things to set up or check after importing:**

- The calendar ID and the Google Sheet (document and tab) are hard-coded to the original author's account. Re-select your own.
- The `sheet` tool's description says "Meeting Log" but it points at a sheet tab named `Sheet3` in a spreadsheet called `Student Details`. Pick the right sheet and make sure its columns are `title`, `date` and `time`.
- The Calendar tool node only maps `Summary` from the AI in the exported JSON. Open it and confirm the **Start** and **End** fields are set (for example, to `$fromAI` values), because the prompt expects the agent to provide them.

### 5. Voice AI Agent

Talk to the assistant with voice notes or text. Replies come back as text **and** as a spoken audio message.

```
                     ┌─ Voice ─▶ Get a file ─▶ STT ─▶ audio_var ─┐
Telegram Trigger ─▶ Switch                                         ├─▶ AI Agent ─┬─▶ Telegram (text reply)
                     └─ Text ──▶ Input_var ──────────────────────┘     │         │
                                                                        │         └─▶ Basic LLM Chain ─▶ TTS ─▶ Telegram1 (audio)
                                          tools: SerpAPI · Gmail · Calculator   (summarise for speech)   (OpenAI)
```

1. **Switch** checks whether the message has a voice note (`message.voice.file_id`) or text.
2. **Voice path:** download the audio file from Telegram, transcribe it with OpenAI (`STT`), and store the transcript in a `text` field. **Text path:** copy the message text into the same `text` field. Both paths feed the same agent.
3. **AI Agent** (`gpt-4o-mini`, per-chat memory) uses a detailed system prompt that tells it to call a tool only when needed:
   - **SerpAPI** for current information, news and web lookups
   - **Gmail** to send an email, in a fixed `Subject / Dear … / Regards` format
   - **Calculator** for any arithmetic
4. The agent's answer goes to Telegram as a text message, and in parallel to a **Basic LLM Chain** that rewrites it as a short, speech-friendly summary. **TTS** (OpenAI text-to-speech) turns that into audio, which `Telegram1` sends back as an audio message.

> Note that the spoken reply is generated for *every* message, including text-only ones. If you only want audio in response to voice notes, add an IF node before the `Basic LLM Chain`.

---

## Getting started

### Prerequisites

- An [n8n](https://n8n.io) instance (cloud or self-hosted). The exports use recent AI Agent node versions (up to `3.1`), so use an up-to-date n8n release.
- An **OpenAI API key**: all five workflows use it.
- Depending on the workflow:

| Service | Used by | Credential type in n8n |
|---------|---------|------------------------|
| Gmail | 1, 2, 4, 5 | Gmail OAuth2 |
| Telegram bot (create one with [@BotFather](https://t.me/BotFather)) | 3, 4, 5 | Telegram API |
| SerpApi ([serpapi.com](https://serpapi.com)) | 3, 4, 5 | SerpApi |
| Google Docs, Calendar, Sheets | 4 | Google OAuth2 (one per service) |

### Import a workflow

1. In n8n, open **Workflows → Add workflow → Import from file** (or paste the JSON).
2. Pick one of the `.json` files in this repository.
3. Open each node that shows a credential warning and **select your own credential**. The exports reference credential IDs from the original instance, which do not exist on yours. No secrets are stored in the files.
4. Update the hard-coded values listed in each workflow section above (email recipient, calendar ID, spreadsheet).
5. Click **Execute workflow** (or send a chat or Telegram message) to test, then toggle the workflow to **Active**.

### Telegram tips

- Telegram triggers need your n8n instance to be reachable over **HTTPS** at a public URL (set `WEBHOOK_URL` when self-hosting, or use a tunnel such as ngrok for local testing).
- A Telegram bot can only have **one webhook at a time**. Workflows 3, 4 and 5 were exported with the same bot credential, so activating more than one with the same bot will make them overwrite each other's webhook. Use a separate bot (and credential) for each, or activate only one at a time.

---

## Concepts these workflows demonstrate

- **AI Agent node** with a chat model, memory and tools (workflows 1, 3, 4, 5)
- **Letting the model fill tool inputs** with `$fromAI(...)` (Gmail, Docs, Sheets, Calendar)
- **Per-user memory** keyed on the Telegram chat id, so conversations don't mix
- **System prompt design** for reliable multi-step tool use (workflow 4) and conditional tool use (workflow 5)
- **Routing** by input type with a `Switch` node and merging branches back into one agent (workflow 5)
- **Speech to text and text to speech** with OpenAI audio (workflow 5)
- **Looping items through an agent, then aggregating** the results (workflow 2)

## Safety notes

- These agents can **send real emails and create real calendar events, docs and sheet rows** on your behalf. Test with a throwaway account or recipient first.
- Anyone who can message your Telegram bot can trigger those actions. Before activating a workflow publicly, add a check on the sender's Telegram user or chat id.
- Review the JSON files before publishing your own fork: they contain account identifiers such as the email address, calendar ID and Google Sheet ID mentioned above. Replace them with placeholders if you don't want them public.
