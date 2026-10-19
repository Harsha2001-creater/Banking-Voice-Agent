# Autonomous Voice Banking Agent (Multi-Agent RAG)

A low-latency, full-duplex conversational voice agent engineered for banking customer support. The system bridges PSTN phone calls through LiveKit SIP trunks, processes audio using Silero VAD and Deepgram Nova-3, and routes caller intents through a LangGraph multi-agent supervisor backed by PostgreSQL (`pgvector` for policy retrieval and `PostgresSaver` for call persistence).

---

## Architecture Overview

```
                  [Inbound PSTN Phone Call]
                             │
                             ▼
                [LiveKit SIP Telephony Gateway]
                             │ (Bi-directional WebSockets)
                             ▼
                 [LiveKit Voice Worker]
                 ├── VAD: Silero (<50ms detection)
                 ├── STT: Deepgram Nova-3 (<250ms)
                 └── TTS: Deepgram Aura / Cartesia Sonic (<200ms)
                             │
             (Streamed Transcripts & User Tokens)
                             ▼
             [LangGraph Multi-Agent Supervisor]
                             │
   ┌─────────────────────────┼─────────────────────────┐
   ▼                         ▼                         ▼
[Accounts Agent]        [Policy RAG Agent]         [Dispute Agent]
├── Balance lookups     ├── pgvector Cosine Search ├── Fraud lock flags
└── Ledger summaries    └── Regulatory PDF chunks  └── HITL escalations
│                         │                         │
└─────────────────────────┴─────────────────────────┘
│
(Streaming LLM Tokens via OpenRouter)
▼
[LiveKit TTS Streamer]
│ (Synthesized Audio Chunks)
▼
[Caller Earpiece]
```

---

## Core Capabilities

- **Sub-800ms End-to-End Latency:** Employs streaming LLM token delivery via OpenRouter directly into real-time TTS audio buffers.
- **Dynamic Turn-Taking & Interruption:** Silero VAD detects user speech during agent output and triggers immediate cancellation of pending TTS audio streams.
- **Agentic Policy RAG:** Retrieves banking compliance guidelines, wire transfer cutoff times, and overdraft fee schedules indexed via pgvector from regulatory manuals.
- **Stateful Call Resumption:** Checkpointed call sessions allow transfer across specialized sub-agents without context loss.
- **Voice-Tuned Output:** Enforces strict phonetic normalization (e.g., "$1,200.50" rendered as "twelve hundred dollars and fifty cents") and restricts responses to under two sentences.

---

## Tech Stack

- **Orchestration & Voice Runtime:** LiveKit Agents SDK (Python)
- **Agent Framework:** LangGraph, LangChain Core
- **Speech Infrastructure:** Deepgram Nova-3 (STT), Deepgram Aura / Cartesia (TTS), Silero (VAD)
- **Model Gateway:** OpenRouter (`meta-llama/llama-3.3-70b-instruct` / `google/gemini-2.5-flash`)
- **Database & Retrieval:** AWS RDS PostgreSQL, `pgvector` extension, `psycopg`
- **Embeddings:** HuggingFace `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions)

---

## Prerequisites

- Python 3.10+
- PostgreSQL instance (v15+) with `vector` extension enabled
- LiveKit Cloud account with an assigned SIP phone number
- API keys: OpenRouter, Deepgram, and LiveKit

---

## Installation & Environment Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/<your-username>/banking-voice-agent.git
   cd banking-voice-agent
   ```

2. **Create a virtual environment:**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Configure Environment Variables:**
   Copy `.env.example` to `.env` and populate all credentials:
   ```bash
   cp .env.example .env
   ```

   ```env
   # LiveKit Telephony & RTC
   LIVEKIT_URL=wss://<project-name>.livekit.cloud
   LIVEKIT_API_KEY=APIxxxxxxxxx
   LIVEKIT_API_SECRET=secretxxxxxxxxx

   # STT / TTS / LLM
   DEEPGRAM_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxx

   # PostgreSQL & Vector Database
   DATABASE_URL=postgresql://postgres:password@<rds-endpoint>:5432/bank_agent
   ```

---

## Database & Vector Initialization

Run the initialization DDL script to create the vector extension, policy chunk table, and account ledgers:

```bash
psql $DATABASE_URL -f src/db/schema.sql
```

Index your regulatory manuals and compliance documentation into vector storage:

```bash
python -m src.rag.ingest --source-dir ./documents/regulatory_manuals
```

---

## Running the Agent

### Local Development (Web Sandbox)

Test speech processing, interruptions, and agent state transitions in your browser via WebRTC:

```bash
python -m src.voice.worker dev
```

Open the LiveKit Agent Sandbox URL printed in your terminal and speak through your local microphone.

### Production Telephony Mode (Inbound SIP)

Run the worker in worker daemon mode:

```bash
python -m src.voice.worker start
```

Point an Inbound SIP Dispatch Rule in the LiveKit Console to this dispatch agent queue name (`banking-voice-agent`). Dial your assigned telephone number to engage the bot.
