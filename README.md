<div align="center">

# 🎙️ Autonomous Voice Banking Agent
### **Full-Duplex Multi-Agent RAG for Real-Time Telephony Support**

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![LiveKit Agents](https://img.shields.io/badge/LiveKit-SIP%20%26%20WebRTC-00E5FF?style=for-the-badge&logo=webrtc&logoColor=black)](https://docs.livekit.io/agents/)
[![LangGraph](https://img.shields.io/badge/LangGraph-Multi--Agent-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white)](https://langchain-ai.github.io/langgraph/)
[![Deepgram](https://img.shields.io/badge/Deepgram-Nova--3%20%26%20Aura-13EF93?style=for-the-badge&logo=deepgram&logoColor=black)](https://deepgram.com/)
[![OpenRouter](https://img.shields.io/badge/OpenRouter-LLM%20Gateway-6366F1?style=for-the-badge)](https://openrouter.ai/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)

<p align="center">
  <em>A production-grade, sub-800ms latency conversational voice system bridging PSTN phone calls via LiveKit SIP trunks to a stateful LangGraph multi-agent supervisor backed by AWS RDS PostgreSQL and pgvector.</em>
</p>

[Architecture](#-system-architecture) •
[Key Features](#-key-features) •
[Tech Stack](#-tech-stack--latency-budget) •
[Quick Start](#-quick-start) •
[Deployment](#-running-the-agent)

</div>

---

## 🏗️ System Architecture

GitHub natively renders the Mermaid diagram below into a visual flow showing the real-time audio and token streaming pipeline:

```mermaid
flowchart TD
    Caller(["📞 Inbound PSTN / WebRTC Caller"]) <-->|"Duplex Audio (SIP / RTP)"| Gateway["🌐 LiveKit SIP Gateway"]
    
    subgraph VoiceWorker ["⚡ LiveKit Voice Worker (Python)"]
        VAD["Silero VAD<br/>(<50ms Barge-in Detection)"]
        STT["Deepgram Nova-3 STT<br/>(<250ms Streaming Transcription)"]
        TTS["Deepgram Aura / Cartesia TTS<br/>(<200ms First-Byte Audio)"]
    end

    Gateway <-->|"Bi-directional WebSocket"| VoiceWorker
    VAD --> STT
    STT -->|"Streamed User Transcript"| Supervisor

    subgraph LangGraph ["🧠 LangGraph Multi-Agent Orchestrator"]
        Supervisor{"🧭 Supervisor Router<br/>(Intent Classification)"}
        
        Accounts["🏦 Accounts Specialist<br/>Balance & Ledger Queries"]
        PolicyRAG["📜 Policy RAG Specialist<br/>Regulatory & Fee Q&A"]
        Disputes["🛡️ Dispute & Fraud Agent<br/>Card Locks & HITL Escalation"]

        Supervisor -->|"Balance / Transfers"| Accounts
        Supervisor -->|"Compliance / Fees"| PolicyRAG
        Supervisor -->|"Fraud / Unrecognized Tx"| Disputes
    end

    subgraph DataLayer ["🗄️ AWS RDS PostgreSQL"]
        Checkpoint[("💾 PostgresSaver<br/>Call Session State")]
        Ledger[("📊 Customer Accounts<br/>& Transactions")]
        VectorDB[("🔍 pgvector Index<br/>Regulatory PDF Manuals")]
    end

    Supervisor <--> Checkpoint
    Accounts <--> Ledger
    PolicyRAG <-->|"384-d Cosine Search"| VectorDB
    Disputes <--> Ledger

    Accounts & PolicyRAG & Disputes -->|"Streaming Tokens via OpenRouter"| TTS
    TTS -->|"Synthesized Audio Chunks"| Gateway
```

---

## ✨ Key Features

| Capability | Technical Implementation |
| :--- | :--- |
| ⚡ **Sub-800ms Turnaround** | Pipes streaming LLM tokens from OpenRouter directly into real-time TTS chunk buffers without waiting for full sentence completion. |
| 🛑 **Natural Barge-In (VAD)** | Silero VAD continuously monitors incoming caller audio (<50ms window) and immediately flushes queued TTS audio when interrupted. |
| 📚 **Agentic Compliance RAG** | Queries `pgvector` HNSW indexes over chunked regulatory PDFs (200–350 tokens) to answer complex fee, wire, and policy inquiries. |
| 🔄 **Stateful Call Checkpointing** | Persists multi-turn conversation threads via `AsyncPostgresSaver`, enabling seamless context handoffs across specialist sub-agents. |
| 🗣️ **Phonetic Normalization** | Enforces strict spoken-word formatting (e.g., rendering `$1,240.50` as *"twelve hundred forty dollars and fifty cents"*) for natural prosody. |
| 🔒 **HITL Fraud Safeguards** | Isolates card-locking and dispute workflows behind human-in-the-loop verification flags and PII regex redaction. |

---

## 🛠️ Tech Stack & Latency Budget

| Layer | Technology | Model / Engine | Target Latency |
| :--- | :--- | :--- | :--- |
| **Telephony & RTC** | LiveKit Cloud & SIP | WebRTC / SIP Trunking | `~40ms` |
| **Voice Activity (VAD)** | Silero | `silero-vad` v5 | `<50ms` |
| **Speech-to-Text (STT)** | Deepgram | `nova-3` (Streaming WebSocket) | `180–250ms` |
| **Agent Orchestration** | LangGraph | `StateGraph` + `AsyncPostgresSaver` | `~15ms` |
| **LLM Brain** | OpenRouter | `meta-llama/llama-3.3-70b-instruct` | `150–250ms TTFT` |
| **Vector Embeddings** | HuggingFace | `all-MiniLM-L6-v2` (384 dims) | `~20ms` |
| **Text-to-Speech (TTS)** | Deepgram / Cartesia | `aura-asteria-en` / `sonic-english` | `150–200ms` |

---

## 📂 Project Structure

```text
banking-voice-agent/
├── .env.example                  # Template for environment credentials
├── .gitignore                    # Git exclusion rules
├── README.md                     # Project documentation
├── requirements.txt              # Python dependencies
├── src/
│   ├── config.py                 # Runtime constants & validated env settings
│   ├── voice/
│   │   ├── worker.py             # LiveKit entrypoint & SIP audio session loop
│   │   ├── adapter.py            # LangGraph-to-LiveKit streaming token bridge
│   │   └── prompts.py            # Phonetic & voice-concise system prompts
│   ├── agent/
│   │   ├── state.py              # BankingVoiceState TypedDict schema
│   │   ├── graph.py              # Compiled StateGraph & Postgres checkpointer
│   │   ├── nodes/                # Supervisor, Accounts, Policy RAG, Disputes
│   │   └── tools/                # RDS SQL queries & pgvector similarity search
│   ├── rag/
│   │   ├── ingest.py             # PDF regulatory manual chunker & vector upsert
│   │   └── embeddings.py         # Local HuggingFace embedding pipeline
│   └── db/
│       ├── session.py            # Async psycopg connection pool
│       └── schema.sql            # PostgreSQL + pgvector DDL schema
└── tests/                        # Routing & retrieval latency unit tests
```

---

## 🚀 Quick Start

### 1. Prerequisites
* **Python 3.10+** installed locally
* **PostgreSQL 15+** (AWS RDS or local) with the `vector` extension enabled
* **LiveKit Cloud** project with an inbound SIP phone number configured
* API keys for **OpenRouter**, **Deepgram**, and **LiveKit**

### 2. Installation

```bash
# Clone the repository
git clone [https://github.com/](https://github.com/)<your-username>/banking-voice-agent.git
cd banking-voice-agent

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 3. Environment Configuration

Copy the example configuration file and populate your credentials:

```bash
cp .env.example .env
```

```ini
# --- LiveKit Telephony & RTC ---
LIVEKIT_URL=wss://<project-name>.livekit.cloud
LIVEKIT_API_KEY=APIxxxxxxxxx
LIVEKIT_API_SECRET=secretxxxxxxxxx

# --- Speech & LLM Gateway ---
DEEPGRAM_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxx

# --- AWS RDS PostgreSQL & pgvector ---
DATABASE_URL=postgresql://postgres:password@<rds-endpoint>:5432/bank_agent
```

### 4. Database & Vector Index Initialization

Initialize the relational tables, `pgvector` extension, and HNSW cosine similarity index:

```bash
psql $DATABASE_URL -f src/db/schema.sql
```

Chunk and embed your regulatory PDF manuals into PostgreSQL:

```bash
python -m src.rag.ingest --source-dir ./documents/regulatory_manuals
```

---

## 🎧 Running the Agent

> [!TIP]
> **Zero-Cost Browser Testing:** Start with **Web Sandbox Mode** to test voice latency, barge-in interruptions, and multi-agent routing using your computer microphone before dialing via carrier telephony.

### Mode A: Local WebRTC Sandbox (Development)

```bash
python -m src.voice.worker dev
```
Open the LiveKit Agent Playground URL output in your terminal to converse with the banking supervisor directly in your browser.

### Mode B: Production Inbound SIP Telephony

```bash
python -m src.voice.worker start
```
1. Navigate to your **LiveKit Cloud Console → Telephony → Dispatch Rules**.
2. Point your inbound US/International SIP number to the worker queue (`banking-voice-agent`).
3. Call the provisioned telephone number from any mobile or landline phone.

---

## 🗺️ Roadmap

- [x] Multi-agent supervisor routing (Accounts, Policy RAG, Disputes)
- [x] AWS RDS `pgvector` integration for regulatory compliance manuals
- [x] Full-duplex interruption handling with Silero VAD
- [ ] Biometric voiceprint caller verification
- [ ] Warm call transfer (`SIP REFER`) to human banking specialists with live summary whisper
- [ ] Real-time Streamlit supervisor observability dashboard

---

<div align="center">
  <sub>Built with ❤️ using LiveKit, LangGraph, OpenRouter, and PostgreSQL</sub>
</div>