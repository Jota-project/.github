# Technical Architecture & Communication

Jota uses a **decentralized event-driven architecture**.

### 1. Communication Protocols
* **WebSockets (Secure):** Used for real-time streaming (Audio from Transcriber, Text from Inference).
* **REST API:** Used for configuration, history retrieval, and authentication.
* **MQTT:** Used for asynchronous events and IoT device signaling.
* **MCP (Model Context Protocol):** Standardized way for the LLM to interact with local/remote tools.

### 2. Data Flow
1. **Input:** `Jota-Transcriber` captures audio -> emits text via MQTT/WS.
2. **Brain:** `Jota-Orchestrator` receives text -> asks `Jota-Inference` for a response.
3. **Action:** If the LLM needs a tool, Orchestrator calls an **MCP Server** (like `jota-db`).
4. **Output:** Response is sent back to the client.

### 🔐 Security Layer
All services implement a **Shared Secret Auth** or **API Key** system managed by `jota-db`.