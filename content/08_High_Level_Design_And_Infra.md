# 8. High-Level Design (HLD) & Infrastructure Details

The architecture decouples the telephony streaming from the backend processing, optimizing for Python/FastAPI asynchronous capabilities while utilizing standard Django toolsets for management, lead ingestion, and the expert resolution dashboard.

### 8.1 System Architecture Diagram

![alt text](../assets/HLD.png)

```mermaid
%%{init: {"layout": "elk","themeVariables":{"spacing":50}}}%%
graph TD
    %% 🟦 Lead Ingestion
    subgraph A[Lead Ingestion]
        classDef indigo stroke:#818cf8,fill:#eef2ff;
        Ads[Google / FB Ads Webhooks]
        AdminUI[Admin Dashboard]
        API_Gateway[API Gateway / Load Balancer]
        Ads --> |API Push| API_Gateway
        AdminUI --> |CSV Upload| API_Gateway
    end
    class A,Ads,AdminUI,API_Gateway indigo;

    %% 🟩 Edge & Telephony
    subgraph B[Edge & Telephony]
        classDef teal stroke:#2dd4bf,fill:#f0fdfa;
        User((Borrower))
        Twilio["📞 Twilio / Exotel SIP Trunk"]
        MediaGateway[FastAPI / LiveKit Gateway Cluster]
        User <--> |PSTN 140-Series| Twilio
        Twilio <--> |WebSockets| API_Gateway
        API_Gateway <--> |WSS Routing| MediaGateway
    end
    class B,User,Twilio,MediaGateway teal;

    %% 🟪 Streaming AI Pipeline
    subgraph C[Streaming AI Pipeline]
        classDef violet stroke:#a78bfa,fill:#f5f3ff;
        STT["🎙️ Deepgram (STT)"]
        TTS["🔊 ElevenLabs (TTS)"]
        MediaGateway --> |Stream Audio Chunks| STT
        TTS --> |Audio Bytes| MediaGateway
    end
    class C,STT,TTS violet;

    %% 🟧 Core Agentic Brain
    subgraph D[Core Agentic Brain]
        classDef orange stroke:#fb923c,fill:#fff7ed;
        CoreApp[Django / LangGraph Orchestrator]
        Decision{"Audio in Cache?"}
        RAG_Engine[RAG Retrieval Engine]
        LLM[GPT-4o-mini / Claude 3.5]
        MediaGateway <--> |Text / Events| CoreApp
        CoreApp <--> |Queries| RAG_Engine
        RAG_Engine --> |Feeds Context| LLM
        CoreApp <--> |Prompts / Tool Calls| LLM

        %% Audio Decision Flow
        CoreApp --> |Request Audio| Decision
        Decision --> |Yes| RedisCache
        RedisCache --> |Return Cached Audio| CoreApp
        Decision --> |No| TTS
        TTS --> |Generate Audio Bytes| CoreApp
        CoreApp --> |Store New Audio| RedisCache
    end
    class D,CoreApp,RAG_Engine,LLM,Decision orange;

    %% 🩷 Async Orchestration
    subgraph E[Asynchronous Orchestration]
        classDef fuchsia stroke:#e879f9,fill:#fdf4ff;
        CeleryWorkers[Celery Job Queue]
        CeleryBeat[Celery Beat Scheduler]
        WhatsApp["💬 WhatsApp Business API"]
        API_Gateway --> |Ingest Leads| CeleryWorkers
        CeleryWorkers --> |Update States| PgVector
        CeleryBeat --> |Dial Commands| Twilio
        CeleryWorkers --> |Webhooks| WhatsApp
    end
    class E,CeleryWorkers,CeleryBeat,WhatsApp fuchsia;

    %% 🟩 HITL Expert Resolution
    subgraph F[HITL Expert Resolution]
        classDef green stroke:#4ade80,fill:#f0fdf4;
        ExpertUser((Human Expert))
        ExpertDashboard[Django Expert UI]
        ExpertUser --> |Use Dashboard| ExpertDashboard
        ExpertDashboard --> |Simulate/Test Output| RAG_Engine
        ExpertDashboard --> |Confirm & Upsert| PgVector
        ExpertDashboard --> |Trigger WhatsApp| CeleryWorkers
    end
    class F,ExpertUser,ExpertDashboard green;

    %% 🩵 Data Layer
    subgraph G[Data Layer]
        classDef cyan stroke:#22d3ee,fill:#ecfeff;
        PgVector[(PostgreSQL + pgvector)]
        RedisCache[(Redis Cache & Audio Index)]
        EFS[(Elastic File Storage - EFS)]
        RAG_Engine <--> |Bank Scripts & FAQs| PgVector
        RedisCache <--> |Read / Write Audio Files| EFS
    end
    class G,PgVector,RedisCache,EFS cyan;

    %% ⬇ Vertical Logical Flow with spacing
    A -->| | B
    B -->| | C
    C -->| | D
    D -->| | E
    E -->| | F
    F -->| | G
```

### 8.2 Component Detailing

1. **Lead Ingestion Engine:**
   * **Ad Platform Webhooks:** Real-time webhooks from Facebook Lead Ads or Google Ads hit the API Gateway and are immediately queued by Celery to insert fresh, high-intent leads into PostgreSQL.
   * **Admin Uploads:** Operations staff can manually upload CSV lead sheets via the Django admin panel for targeted campaign dialing.

2. **Load Balancer & Media Gateway Cluster:** 
   * An AWS Application Load Balancer (or NGINX/HAProxy) distributes incoming WebSocket connections from the SIP Provider (Twilio/Exotel) across a horizontally scaling cluster of FastAPI/LiveKit Media Gateway pods. This ensures high availability and prevents CPU bottlenecks during audio processing.

3. **Multi-Tier TTS Caching (Redis + EFS):**
   * To achieve ultra-low latency, the system intercepts LLM text at sentence boundaries and checks Redis for an MD5 hash. Instead of storing heavy raw audio bytes in memory, **Redis acts as a fast lookup index returning a file path**. The Media Gateway then instantly pulls the `.wav` file from **Elastic File Storage (EFS)**—a local network mount shared across all gateway pods. If it's a cache miss, it calls ElevenLabs, streams to the user, and asynchronously saves the file to EFS while updating the Redis pointer.

4. **The Agentic Brain (Django/FastAPI):**
   * Manages the conversation state machine using LangGraph or CrewAI. It holds conversational context, tracks the `agentic_callback_count` escalation triggers, and executes Tool-Calling logic for deterministic RAG retrieval.

5. **HITL Expert Dashboard & Pre/Post Testing Simulator:**
   * When the AI encounters an unseen query and pivots, a ticket is generated in the Expert Dashboard.
   * **Testing Simulator:** The dashboard features a unique testing interface. Before pushing a new canonical answer to the database, the expert types the answer and clicks "Simulate". The dashboard runs the user's original query against the RAG/LLM pipeline in a sandbox, showing the expert exactly how the AI will respond with the new context.
   * **Resolution & Follow-up:** Once satisfied that the AI responds perfectly, the expert clicks "Approve." The system automatically embeds and upserts the knowledge to `pgvector`, triggers the WhatsApp response to the user, and schedules the AI to call the user back to resume the conversation.

6. **Database (PostgreSQL + pgvector):**
   * Acts as the primary transactional datastore for leads and system state, as well as the robust vector database for RAG embeddings.
