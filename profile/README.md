# 🤖 Jota-Project: The Next-Gen Open AI Assistant
> **A modular, high-performance ecosystem blending LLM intelligence with real-world Home Automation.**

Jota is not just a chatbot; it's a **distributed brain**. Built with a microservices architecture, it bridges the gap between state-of-the-art Large Language Models and your physical environment using **MCP (Model Context Protocol)**, **MQTT**, and **WebSockets**.

### 🏗️ Ecosystem Architecture
| Service | Language | Role | Status |
| :--- | :--- | :--- | :--- |
| **[Orchestrator](https://github.com/jota-project/jota-orchestrator)** | Python | Logic, Tool Use & Routing | `Pre-Alpha` |
| **[Inference](https://github.com/jota-project/jota-inference)** | C++ | Local LLM Execution (llama.cpp) | `In-Dev` |
| **[Transcriber](https://github.com/jota-project/jota-transcriber)** | C++ | Real-time Whisper Transcription | `Functional` |
| **[Database](https://github.com/jota-project/jota-db)** | Python | Persistence, Memory & Auth | `Functional` |

### 🌟 Key Features
* **Performance First:** Core heavy-lifting (Inference/Audio) written in C++.
* **Privacy-Centric:** Designed to run 100% locally.
* **Extensible:** Easy to add new "Skills" via MCP or new Clients (Voice, Web, Mobile).
* **Enterprise-Ready:** Built-in Auth, Connection Limiting, and Metrics.

---
*Built for the community, powered by you.*
