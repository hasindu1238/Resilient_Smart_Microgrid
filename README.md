# Rural Health Center AI Energy Prioritizer: The Village Micro-grid Brain


## Project Summary & Real-World Context
In rural Sri Lanka, grid instability is not a mere inconvenience; it is a threat to life. As highlighted in Sustainable Development Goal 3 (Good Health & Well-being), the integrity of temperature-sensitive vaccines and medicines is paramount. This project implements an intelligent, agentic controller for village micro-grids that serves as the "Brain" to the IoT "Nervous System." 


By utilizing a multi-agent framework, we automate the "Team of Engineers" (Researcher, Engineer, Tester, Designer) to maintain a continuous learning loop. The system senses power drops, predicts battery depletion, optimizes load-shedding based on a strict 4-level hierarchy, and acts autonomously to protect the medical cold chain.


## Tech Stack
- **Language:** Python 3.10+
- **API Framework:** FastAPI (High-performance telemetry ingestion)
- **Agent Framework:** LangChain / AutoGen (Multi-Agent orchestration)
- **Data Integrity:** Pydantic (Strict schema validation)

- **Frontend/UI:** Streamlit (Real-time XAI decision dashboard)
- **Communication:** MQTT & HTTP (Optimized for ESP32/WiFiClientSecure)
- **Containerization:** Docker & Docker Compose (Edge-gateway ready)


## Installation & Setup


### Local Development (Python venv)
1. Clone the repository:
   ```bash
   git clone https://github.com/vinu-labs/village-energy-prioritizer.git
   cd village-energy-prioritizer
