# 🚀 AI Content Pipeline (LinkedIn Posts)
**🎥 Watch the System Architecture & Demo (Loom) https://www.loom.com/share/c85dba6330eb4f21a6c22b9330006235**

## System Overview
A semi-automated, state-driven content ingestion and drafting pipeline built in n8n. The system monitors tech-focused RSS feeds, scores candidates based on custom heuristics, drafts brand-aligned LinkedIn posts using Llama 3.3, generates contextual visual assets via Gemini/Cloudinary, and halts for human approval via an interactive Slack UI.

![Slack UI Screenshot](https://github.com/user-attachments/assets/1b14447b-a85d-4e18-95e8-5a955c176bed) 

* The interactive "Human Gatekeeper" UI built with Slack Block Kit.*

---

## 🎯 Addressing Core Constraints

### 1. The Human Gatekeeper
**Requirement:** *No content should ever go live without explicit, manual approval.*
* **Implementation:** The generation pipeline intentionally halts after pushing an interactive Block Kit payload to a private Slack channel. The database state is locked to `pending_approval`. A separate interactive webhook router (Workflow 2) listens for the reviewer's decision to either `Approve & Post`, `Regenerate`, or fetch the `Next Article`. 

### 2. Memory & Prevention
**Requirement:** *Ensure no duplicate content is ever generated or posted.*
* **Implementation:** An independent PostgreSQL database manages the state machine. 
    * **Ingestion Phase:** Incoming RSS URLs are cross-referenced against the DB. Only net-new URLs are ingested.
    * **Approval Phase:** Articles move through strict state transitions (`candidate` -> `rejected` -> `pending_approval` -> `posted`). Once an article is explicitly rejected or posted, it is permanently locked out of the queue.

### 3. Brand DNA
**Requirement:** *Capture specific voice, cadence, and formatting quirks.*
* **Implementation:** The "Style Guide" is enforced at the LLM configuration layer. The Groq (Llama 3.3 70B) system prompt acts as the brand governor, enforcing the Brand persona. It contains explicit instructions regarding hook structures, mandatory paragraph spacing (for LinkedIn readability), and a prohibition on generic AI emojis, ensuring the digital intern sounds natively human and on-brand.


![n8n Canvas Screenshot](https://github.com/user-attachments/assets/d85bf234-ac51-4a75-b1a4-6119646837d9)
![n8n Canvas Screenshot2](https://github.com/user-attachments/assets/bbd15345-3418-436a-9e8a-2586a4669ff1)

* The ingestion, scoring, and drafting and rounting workflows and pipeline.*

---

## 🏗️ Engineering Evaluation Criteria

### Security Mindset (Credential Handling)
* **n8n Credential Manager:** No API keys, database passwords, or auth tokens exist in plaintext within the workflow logic. Everything is routed through n8n’s native credential manager.
* **HTTP Node Security:** For custom API calls (like Slack updates or mock endpoints), I utilized n8n's **Predefined Credentials** feature. This ensures that when the JSON workflows are exported and committed to this public repository, there is zero risk of credential leakage or security vulnerabilities.

### Logical Resilience (Edge Cases & Fallbacks)
* **Self-Healing Ingestion:** If the primary RSS feed returns 0 items for the current 24-hour window (slow news day), the system dynamically expands the timeframe to fetch yesterday's top articles, ensuring the pipeline never runs dry.
* **LLM Hallucination Bypasses:** Modern LLMs often struggle with strict JSON function calling, occasionally injecting markdown backticks that crash standard parsers. I implemented a custom Regex parsing node (The "Titanium Hammer") to aggressively extract and clean JSON payloads, guaranteeing 100% parse reliability.
* **Two-Phase Commits:** When posting, the database status only updates to `posted` *after* the API returns a `200 OK` success response. This prevents "zombie" records where the database thinks a post is live, but the API actually failed.

### Modular Design
* **Separation of Concerns:** The system is divided into logical sub-domains. Workflow 1 handles Cron-triggered ingestion and drafting. Workflow 2 acts purely as an API router/controller listening for Slack interactions. 
* **Environment Boundaries (API Mocking):** Due to LinkedIn's standard 72-hour OAuth verification window for new Company Pages, I chose not to compromise my personal profile for testing. Instead, I built a simulated mock endpoint (Workflow 3) to receive the final production payload. This validates payload integrity (Text, Source URL, Image CDN URL) while strictly maintaining environment boundaries.

---

## 📁 Repository Contents
* `/workflows/wf1_ingestion_and_drafting.json` - The main Cron-triggered pipeline.
* `/workflows/wf2_interactive_router.json` - The Slack webhook listener and CRUD operator.
* `/workflows/wf3_mock_production_api.json` - The simulated LinkedIn endpoint.
* `/workflows/wf4_error_workflow_trigger.json` - The Error Notifyer Workflow.

*Note: To review the logic, you can import these JSON files directly into any n8n instance.*
