# 3. Highlighted Features (System Differentiators)

What sets the AI Money Lender Call System apart from standard voice bots are these proprietary safety, caching, and learning mechanisms:

1. **Continuous Learning RAG (The "I'll Check" Protocol)**  
   * **The Problem:** Bots often hallucinate financial advice or hit a frustrating "I don't know" dead end.
   * **The Solution:** If the RAG retrieval confidence is low (<0.85), the agent safely pivots and gives the user control: *"I want to be 100% accurate on that policy. Would you like me to transfer you to a human agent right now, or should I verify this with my team and call you back later?"* If the user chooses a callback, the unseen query generates a ticket. A human expert answers it via a Django Admin portal, which sends a WhatsApp to the user AND automatically formats, embeds, and upserts the new Q&A into `pgvector`—ensuring the AI knows the answer for the next caller. If they choose an immediate transfer, the call is instantly routed via SIP to the live customer care queue.

2. **The 3-Strike Escalation Matrix**  
   * **The Problem:** Users get frustrated when called repeatedly by a bot that doesn't remember previous context or fails to resolve issues across multiple touchpoints.
   * **The Solution:** A global `agentic_callback_count` tracks every interaction (Calls and WhatsApps) per lead. If automated attempts fail to resolve the user's intent 3 times, the AI is locked out. The lead is pushed to a "High-Touch" dashboard where a human closer sees the entire timeline and manually calls, picking up exactly where the AI left off.

3. **Multi-Tier Audio Caching Architecture for Ultra-Low Latency**  
   * To achieve sub-10ms response times and drastically reduce TTS API costs (e.g., ElevenLabs), the system proposes multiple caching strategies to be evaluated during implementation:
     * **Option A (Exact-Match TTS Caching):** Bypassing the TTS API by matching MD5 hashes of the LLM text. To avoid hogging expensive Redis memory with raw audio bytes, Redis only stores the string pointer/URL to the audio file. We have two sub-options for the audio storage layer:
       * **Option A-1 (EFS / Local Network Mount):** The audio files are stored in Elastic File Storage (EFS) mounted directly on the media server. This provides near-zero latency disk I/O without the network overhead of S3.
       * **Option A-2 (S3 + CDN):** The audio files are stored in AWS S3 and fetched via a globally distributed CDN (e.g., CloudFront). This ensures extremely fast edge fetches if the media gateways are distributed across multiple regions.
     * **Option B (Semantic Caching):** Bypassing the LLM entirely by embedding the user's incoming STT transcript (e.g., "What is the rate?") and performing a vector similarity search (e.g., Cosine > 0.95) against previously answered queries. If matched, the system returns the exact historical text, instantly guaranteeing an Exact-Match TTS cache hit.

4. **Deterministic RAG via Tool-Calling**  
   * To maximize the TTS cache hit rate, the LLM is restricted to Tool-Calling for factual answers (e.g., `answer_policy_question()`). This returns hardcoded strings instead of free-form generated text, preventing financial hallucinations and guaranteeing a 100% TTS cache hit for FAQs.
