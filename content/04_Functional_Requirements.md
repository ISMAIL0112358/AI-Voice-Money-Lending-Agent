# 4. Functional Requirements (FRs)

- **FR1 (Autonomous Dialing):** The Campaign Manager must fetch leads from the database and dial via a 140-series promotional SIP trunk at a controlled rate (e.g., 1.6 Calls Per Second) to prevent carrier spam filtering.
  * **Potential Tools:** Celery/Kafka (Job Queue), PostgreSQL (Database), Twilio/Exotel (SIP Trunking).

- **FR2 (Conversational Audio Pipeline):** The system must handle real-time WebSockets, Voice Activity Detection (VAD), and barge-in (interruption) handling to ensure natural conversational turn-taking.
  * **Potential Tools:** LiveKit or FastAPI (WebSocket Gateway), Deepgram Nova-3 (STT), ElevenLabs (TTS).

- **FR3 (RAG & Knowledge Retrieval):** The system must query a Vector DB to accurately answer specific loan policy questions using the Tool-Calling pattern.
  * **Potential Tools:** LangGraph / CrewAI (State Machine), pgvector / Pinecone (Vector Database), GPT-4o-mini / Claude 3.5 (LLM).

- **FR4 (HITL Handoff):** The system must route low-confidence queries (Score < 0.85) to an internal queue, trigger a WhatsApp callback response from a human expert, and auto-ingest the reply back into the RAG database.
  * **Potential Tools:** Django Admin / Internal React Portal (Ticket UI), Twilio / Gallabox (WhatsApp API).

- **FR5 (Escalation Rule):** If a user undergoes 3 automated interactions, the system must abort further automated dials and route the lead context to a human queue.
  * **Potential Tools:** Django ORM / Redis (Interaction Counters), Sales Dashboard (Escalation UI).

- **FR6 (Observability & Logging):** Every call must generate a distributed system trace, a dedicated LLM execution trace (for prompt paths, tool calls, and token usage), a turn-by-turn text ledger, and a dual-channel stereo audio recording stored in the cloud for debugging.
  * **Potential Tools:** LangSmith / AgentOps (LLM Orchestration Tracing), Signoz / OpenTelemetry (System/Network Tracing), Prometheus / Grafana (Metrics), AWS S3 (Stereo Audio Storage).
