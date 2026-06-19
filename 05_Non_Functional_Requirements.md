# 5. Non-Functional Requirements (NFRs)

- **NFR1 (Latency):** Total Audio-to-Audio response time (P99) must be < 1.2 seconds. Time-to-First-Token (TTFT) from the LLM must be < 400ms.
- **NFR2 (Scale & Concurrency):** The system must effortlessly handle the target 1000 calls/hour (approx 2-5 concurrent streams) utilizing a single Media Gateway, with load testing proven at 2x scale (2000 calls/hour) for safety overhead.
- **NFR3 (Compliance):** The system must operate strictly between 11:00 AM and 5:00 PM, scrubbing all leads against the TRAI DND (National Customer Preference Register) registry.
- **NFR4 (Performance):** The TTS audio cache must achieve a >40% hit rate on standard state-machine prompts.
