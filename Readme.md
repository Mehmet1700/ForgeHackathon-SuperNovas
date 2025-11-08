# Super Nova — Telegram LLM Chatbot (n8n)

**Status:** Hackathon prototype (mix of working, partial, and draft workflows)  
**Team:** Super Nova
**Hackathon:** Forge Hackathon 2024
**Members:** Carlota , Libero, Mehmet

## TL;DR
A Telegram-first customer-support chatbot in n8n. Users message (voice or text), we transcribe audio, keep short-term memory, query an LLM to extract intent + slots (PolicyID, action), route the flow, validate against Airtable, and respond. Extras include an address-existence check via a maps API (prototype) and a sentiment analysis spike (prototype).

---

## What’s in this folder
Each JSON file is an importable n8n workflow:

1) **First workflow.json** — end-to-end baseline  
   - Telegram → speech-to-text (for voice) → Gemini → ask for PolicyID → route to **ChangeAddress** / **ClaimPolicy** → Airtable check → reply.  
   - Contains LLM message node, ElevenLabs STT, a `Switch` on model output, and Airtable lookup for `Policies`.
2) **First workflow copy 2.json** — baseline + memory + safer parsing  
   - Adds a **Simple Memory** buffer (per chat id) and a **JSON to Variables** Code node that extracts `{PolicyID, Function}` even if the LLM output isn’t strict JSON.  
   - Includes positive/negative Telegram replies based on whether the policy is found in Airtable.
3) **First workflow carlota.json** & **First workflow carlota (1).json** — address validation POC  
   - Adds a Python Code node preparing an example address and an HTTP Request to **OpenStreetMap Nominatim** to check if it exists; then a small LLM check for “exists/doesn’t exist”.  
   - Demonstrates a **Wait** node pattern (e.g., ask → pause → resume). 
4) **Fixing the loop.json** — sub-workflow “brain”  
   - **JSON COLLECTOR** agent prompt to normalize user messages to a minimal JSON schema:  
     ```json
     { "PolicyID": "POLICY_ID_HERE", "Cause": "ChangeAddress|ClaimPolicy" }
     ```
   - Conversational memory (small context window), Airtable upsert for session state (`resumeUrl`), guards for valid/invalid policies, and an “ask for missing values” agent turn. 
5) **Fixing the loop – Router.json** (+ **copy**) — orchestrator  
   - Merges voice/text paths, looks up the session in Airtable, posts back to a `resumeUrl` if present, and/or triggers the **Fixing the loop** sub-workflow.
6) **sentiment analysis v1.json** — sentiment spike  
   - Telegram + audio/text handling skeleton intended to host a sentiment scorer to adapt the persona/tone. (Not wired into the main router yet.)


---

## Architecture (high level)

**Ingress**  
- Telegram Trigger → detect **voice** vs **text** → Merge into a normalized message.

**Voice path**  
- Get Telegram voice file → **ElevenLabs** speech-to-text → normalized text.

**LLM + Memory**  
- Small **Buffer Memory** (keyed by chat id) keeps the last turns.  
- **LLM (Gemini 2.x)** used in two ways:
  1) *Ask/Respond* (e.g., ask for missing PolicyID),
  2) *Extract/Normalize* (emit `{PolicyID, Function/Cause}`).

**Routing + Data**  
- `Switch` node decides the branch: `PolicyID` missing → ask again; `ChangeAddress` or `ClaimPolicy` → continue.  
- **Airtable**: validate `Policy ID` and read policy metadata; store/update session and `resumeUrl` for wait/resume flows.

**Extras (POC)**  
- **Map API** via OpenStreetMap Nominatim to check whether a user-provided address exists.  
- **Sentiment**: planned hook to adjust answer tone and escalation rules.

---

## Features

- **Telegram, voice & text**: Works for both input modes; voice is auto-transcribed to text first.  
- **Hot-swappable LLM**: Using Gemini today; swapping models only requires updating the LLM node/model id.  
- **Short-term memory**: Keeps a bounded window per chat to avoid re-asking.  
- **Intent + slot extraction**: Normalizes user input into `{PolicyID, Function/Cause}` with fallback parsing when the LLM returns partial/loose JSON.  
- **Two primary flows** (shipping today): **Change Address** and **Claim Policy**.  
- **Address existence check (POC)**: Calls Nominatim to verify if an address likely exists.  
- **Wait/Resume orchestration**: Ask a question, pause, store `resumeUrl`, and resume when the user answers.  
- **(Draft) Sentiment hook**: Endpoint to compute sentiment and influence style/escalation.

---

## Setup

### Prerequisites
- n8n (Cloud or Self-hosted)
- Telegram Bot & token
- API keys/tokens for:
  - **Google Gemini (PaLM)**
  - **ElevenLabs** (speech-to-text)
  - **Airtable** (Personal Access Token)

### Credentials referenced in the workflows
- `Telegram account`
- `Google Gemini(PaLM) Api account`
- `ElevenLabs account`
- `Airtable Personal Access Token account`

### Airtable
- Base: your InsurTech/Hackathon base
- Example tables used:
  - `Policies` — for validating `Policy ID`
  - `Session` — for storing chat state and `resumeUrl`

> You can import every `.json` via **n8n → Import from file**. After import, open each workflow and assign your own credentials.

---

## Running the demo

1. **Import** `First workflow.json` (baseline) and **assign credentials**.  
2. In Telegram, send a text or voice message.  
3. The bot will:
   - Transcribe voice (if any),
   - Ask for **PolicyID** if missing,
   - Route to **ChangeAddress** / **ClaimPolicy** based on your message,
   - Validate the policy in Airtable and answer accordingly.

**Address check (POC):** Import `First workflow carlota.json`, set the HTTP Request node to Nominatim (or your map API), and replace the sample address with user-extracted values.

**Loop orchestration:** Import `Fixing the loop – Router.json` and `Fixing the loop.json`, then connect your Airtable `Session` table. The router will find an existing `resumeUrl` and resume a pending question, or call the sub-workflow to collect/ask for the missing fields.

---

## Message → JSON contract

We normalize user input to the minimal JSON below.  
Downstream nodes assume **either** a missing `PolicyID` (ask again) **or** one of the two actions:

```json
{ "PolicyID": "123456", "Function": "ChangeAddress|ClaimPolicy" }
